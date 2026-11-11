# DoorDash Delivery Time Prediction

A beginner-friendly regression project where I explored food-delivery data, compared prediction models, and investigated what is associated with longer delivery times.

> This is an educational project based on a DoorDash-style business scenario. It is not an official DoorDash project and is not affiliated with DoorDash.

## Project Dashboard Preview

![DoorDash delivery time prediction dashboard preview](Overview%20of%20Delivery%20Time%20Prediction-DoorDash.png)

*Static visual preview for the project. It is a design mockup, not an exported screenshot of a completed interactive Power BI report.*

## The problem

Food-delivery apps show an estimated arrival time when a customer places an order. An estimate that is too optimistic can lead to disappointed customers, refunds, and poor reviews.

For this project, I took the role of a data scientist on a food-delivery logistics team. The goal was to predict delivery time in minutes using information available when an order is placed, evaluate the predictions honestly, and explain which factors the selected model relies on.

## What I worked on

I completed the project in stages:

1. Explored the training data: column types, missing values, duplicates, delivery-time distribution, outliers, and relationships between features and the target.
2. Cleaned working copies of the datasets and documented how missing values and unusually long delivery times were handled.
3. Created time-based features from the order timestamp and an interaction between delivery distance and restaurant preparation time.
4. Used a chronological training/validation split to simulate predicting later orders from earlier orders.
5. Built mean and median baselines, then trained Linear Regression and Random Forest regression models.
6. Compared the models using Mean Absolute Error (MAE), checked training-versus-validation performance, and inspected prediction errors.
7. Used permutation importance and grouped error analysis to understand what the selected model relied on and where it struggled.
8. Retrained the selected model using all labelled training data and generated predictions for the held-out test orders.

## Dataset and privacy

The supplied files contained:

- **Training set:** 8,000 orders, including `delivery_minutes`.
- **Test set:** 2,000 orders, without the target column.
- **Time period:** March 1 to August 31, 2025.

The original CSV files and generated prediction file are not included in this public repository because the datasets are subject to a sharing restriction. They remain on my computer.

To run the notebook locally, place the supplied files in a local `data/` folder with these names:

```text
data/
├── deliveries_train.csv
└── deliveries_test.csv
```

The `.gitignore` file excludes the entire `data/` and `outputs/` folders so these files are not accidentally committed.

## Exploratory analysis: what stood out

Some findings from the training data:

- Average delivery time was **32.05 minutes**, and the median was **31 minutes**.
- The target was right-skewed, with skewness of approximately **2.61**.
- The IQR rule flagged **146 orders (1.82%)** above the 55-minute upper boundary. I investigated these values rather than automatically removing them, because very long deliveries may be genuine operational delays.
- Distance had the strongest numerical correlation with delivery time (**0.53**), followed by restaurant average preparation time (**0.41**).
- `items_count` and `order_subtotal` were strongly correlated with each other (**0.91**), which is worth considering when interpreting model inputs.
- Heavy-rain and suburban-zone orders had higher average delivery times in the training data. These are observed associations, not proof of causation.

## Feature engineering

I extracted the following features from `order_placed_at`:

- `order_hour`
- `day_of_week`
- `order_month`
- `is_weekend`
- `time_period` (Morning, Afternoon, Evening, Late Evening)

I also created `distance_prep_interaction` by multiplying `distance_km` by `restaurant_avg_prep_minutes`.

The final model used 14 candidate input features:

**Numerical:** `restaurant_avg_prep_minutes`, `distance_km`, `items_count`, `order_subtotal`, `order_hour`, `order_month`, `is_weekend`, `distance_prep_interaction`.

**Categorical:** `restaurant_id`, `cuisine`, `city_zone`, `weather`, `day_of_week`, `time_period`.

I excluded `order_id` because it is an identifier. I also excluded `courier_vehicle` and `courier_trips_completed` because it was not established that these courier details would be available at the moment an order is placed. `delivery_minutes` was used only as the target.

## Validation approach

I sorted the labelled data by order timestamp and split it by calendar date:

- **Training:** 6,368 orders, March 1 to July 25, 2025.
- **Validation:** 1,632 orders, July 26 to August 31, 2025.

Keeping all orders from a calendar date in the same partition avoided splitting one date across both sets. This chronological split better reflects predicting future orders using historical data.

I used **Mean Absolute Error (MAE)** as the main evaluation metric. MAE is measured in minutes, so it is straightforward to interpret: lower is better.

## Model results

| Model | Validation MAE |
|---|---:|
| Mean baseline | 7.6338 minutes |
| Median baseline | 7.5809 minutes |
| Linear Regression | **3.9573 minutes** |
| Random Forest Regressor | 4.0926 minutes |

Linear Regression was the best of the models I tested. Its validation MAE was approximately **47.8% lower** than the median baseline.

Random Forest had a lower training MAE (**2.165 minutes**) than validation MAE (**4.0926 minutes**), suggesting a larger generalisation gap. Linear Regression's training MAE was **4.179 minutes**, compared with **3.9573 minutes** on validation. A slightly lower validation error can happen when the later period is somewhat easier to predict; it is not, by itself, evidence of a problem.

## What mattered to the selected model?

I used permutation importance on the validation data. The largest average increases in MAE after shuffling a feature were:

| Feature | Increase in validation MAE |
|---|---:|
| `distance_km` | 4.4752 minutes |
| `restaurant_avg_prep_minutes` | 2.9530 minutes |
| `items_count` | 0.3637 minutes |
| `weather` | 0.2490 minutes |
| `time_period` | 0.1430 minutes |

This suggests that the model relied most on delivery distance and the restaurant preparation estimate. Permutation importance measures how much this particular model's predictions depend on a feature; it does not establish causation. Correlated features can also share predictive information.

## Where did the model struggle?

The overall validation MAE was good, but the errors were not evenly distributed:

| Actual delivery-time group | Validation orders | MAE |
|---|---:|---:|
| Up to 20 minutes | 164 | 4.44 minutes |
| 21–40 minutes | 1,181 | 3.33 minutes |
| 41–55 minutes | 263 | 4.64 minutes |
| Above 55 minutes | 24 | **23.88 minutes** |

The model had its largest difficulty with very long deliveries. For the 24 validation orders above 55 minutes, average actual time was **70.83 minutes**, while average prediction was **47.12 minutes**. The largest absolute validation error was **96.13 minutes**.

Across validation orders, **72.30%** of predictions were within 5 minutes and **96.38%** were within 10 minutes. Average signed error was close to zero (**−0.06 minutes**), but that overall average hides the model's tendency to underestimate some very long deliveries.

The 55-minute grouping was based on the training target's IQR outlier boundary. It is an exploratory analysis threshold, not a verified DoorDash late-delivery service standard.

## Final test predictions

After selecting Linear Regression, I retrained the pipeline using all **8,000 labelled training records** and generated predictions for the **2,000 held-out test orders**.

The output was saved locally as:

```text
outputs/predictions.csv
```

It contains two columns:

- `order_id`
- `predicted_minutes`

The saved file passed the checks performed in the notebook: 2,000 rows, the two expected columns, no missing values, no duplicate order IDs, and no negative predictions.

The test set does not contain actual delivery times, so a test MAE could not be calculated. **3.9573 minutes is the chronological validation MAE, not a measured test-set score.**

## Tools and libraries

- Python
- Jupyter Notebook
- Pandas and NumPy
- Matplotlib and Seaborn
- Scikit-learn

## Repository structure

```text
Delivery_Time_Prediction/
├── data/                         # Local datasets; not committed
├── notebooks/
│   └── 01_data_exploration.ipynb
├── outputs/                      # Local predictions; not committed
├── Overview of Delivery Time Prediction-DoorDash.png
├── .gitignore
├── README.md
└── requirements.txt
```

## Run it locally

1. Clone this repository.
2. Place the supplied CSV files in the local `data/` folder. They are intentionally not included in this public repository.
3. Install the dependencies:

   ```bash
   pip install -r requirements.txt
   ```

4. Start Jupyter Notebook:

   ```bash
   jupyter notebook
   ```

5. Open `notebooks/01_data_exploration.ipynb` and run the cells in order.

## Limitations and what I would improve next

This was a learning project, so there are a few things I would check before using a model like this in a real delivery operation:

- Confirm that `restaurant_avg_prep_minutes` is a historical estimate available when the order is placed, and that `weather` reflects information available at that same time. If either uses information recorded later, it should be removed or rebuilt to avoid leakage.
- Investigate very long deliveries further. They are important cases, and the current Linear Regression model underestimates them.
- Try additional models and carefully tune them using time-based validation, while keeping the final test set untouched.
- Validate over multiple time windows to check whether model performance is stable over time.
- Compare errors across zones and weather categories using larger samples before making operational recommendations.

## A note

I built this project to practise the complete workflow of a regression problem—from understanding raw data to evaluating a model and producing predictions. My biggest takeaway was that a good MAE is only part of the work: understanding the errors and being honest about what the model can and cannot tell us matters just as much.
