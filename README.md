# "Beyond the Label: Are 'Healthy' Recipes Really Healthy?"
### To what extent do semantic health labels (e.g., “low fat”, “gluten free”, “healthy”) embedded in recipe names and tags correspond to quantified nutritional values, and how do these discrepancies influence user-assigned ratings?

by Sunny Sun (chs019@ucsd.edu)

## Introduction

I like food and I like cooking, and I’m also really into staying on a healthy diet—so I’m pretty familiar with nutrition data. That’s why I’ve always wondered if recipes labeled as “healthy” actually live up to their name. Do these labels reflect real nutritional value, or are they just used to make recipes sound better? In this project, I explore that question using the Recipes and Ratings dataset from Food.com.

My main research question is: **Are recipes labeled as “healthy” actually healthier based on nutrition facts, and does that label influence user ratings?** To answer this, I created a Healthy Index using normalized nutrition values, compared labeled vs. unlabeled recipes, and predicted user ratings based on both nutrition and label metadata.

The dataset contains about 230,000 recipes and includes metadata such as prep time, ingredient count, nutrition info (in %DV), and user-submitted ratings. Key columns include:
- `tags`: list of descriptive labels (e.g. “gluten free”, “healthy”)
- `nutrition`: [calories, total fat, sugar, sodium, protein, saturated fat, carbs] in %DV
- `name`: recipe title
- `average_rating`: average user rating (0–5)


## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

To prepare the dataset for analysis, I:

- Merged the recipe metadata with user ratings to compute an `average_rating` column per recipe.
- Parsed the `nutrition` column into 7 numeric columns: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `sat_fat`, and `carbs`.
- Normalized the nutrition values to account for serving size inconsistencies.
- Created health-related label flags (`healthy`, `low_in_something`, `free_of_something`) based on the presence of keywords in `tags` and `name`.
- Constructed a custom `healthy_index` that rewards protein and penalizes fat, sugar, sodium, and carbs.

Below is a preview of the cleaned DataFrame:

```markdown
| name                                 |   year |   healthy_index |   average_rating | healthy   | low_in_something   | free_of_something   |
|:-------------------------------------|-------:|----------------:|-----------------:|:----------|:-------------------|:--------------------|
| 1 brownies in the world    best ever |   2008 |     -0.755058   |                4 | False     | False              | False               |
| 1 in canada chocolate chip cookies   |   2011 |     -0.700723   |                5 | False     | False              | False               |
| 412 broccoli casserole               |   2008 |     -0.00513347 |                5 | False     | False              | False               |
| millionaire pound cake               |   2008 |     -0.769099   |                5 | False     | True               | False               |
| 2000 meatloaf                        |   2012 |     -0.0430712  |                5 | False     | False              | False               |
```



### Univariate Analysis

The plot below shows the distribution of `average_rating`. Ratings tend to skew toward the high end, with most falling between 4 and 5.

<iframe
  src="assets/univariate_rating_distribution.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>



### Bivariate Analysis

The following boxplot shows how the presence of health-related labels affects user ratings. While labeled recipes have slightly lower median ratings, the difference is small.

<iframe
  src="assets/rating_by_label_boxplot.html"
  width="800"
  height="550"
  frameborder="0"
></iframe>

This scatter plot explores the relationship between `healthy_index` and `average_rating`. A slight negative correlation suggests healthier recipes may receive marginally lower ratings.

<iframe
  src="assets/healthy_index_vs_rating_scatter.html"
  width="800"
  height="550"
  frameborder="0"
></iframe>



### Interesting Aggregates

The table below shows average healthy index values for recipes grouped by label category:

```markdown
|                   | low_in_something    | low_fat             | low_carb           | healthy           | gluten_free       |
|:------------------|:--------------------|:--------------------|:-------------------|:------------------|:------------------|
| healthy           | 14564/31727 (45.9%) | 10884/17363 (62.7%) | 5069/26136 (19.4%) | nan               | nan               |
| free_of_something | 1687/32837 (5.1%)   | 858/15622 (5.5%)    | 736/18702 (3.9%)   | 1086/19899 (5.5%) | 2301/4677 (49.2%) |
| gluten_free       | 800/31484 (2.5%)    | 359/13881 (2.6%)    | 390/16808 (2.3%)   | 443/18302 (2.4%)  | nan               |
| low_fat           | 11871/29915 (39.7%) | nan                 | nan                | nan               | nan               |
| low_carb          | 14829/29915 (49.6%) | 4751/21949 (21.6%)  | nan                | nan               | nan               |
```

Recipes labeled as "healthy" have a slightly better (less negative) healthy index, but slightly lower average ratings. This may reflect a gap between nutritional quality and user preferences.

We also tracked average healthy index by year:

<iframe
  src="assets/healthy_index_trend_by_year.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

## Assessment of Missingness

### NMAR Analysis

After inspecting the dataset, I found that the `description` column contains missing values for a small number of recipes. I believe this missingness is likely **Not Missing At Random (NMAR)**.

Here's my reasoning: if a recipe has no description, it may be because the contributor chose not to provide one. This decision could be influenced by factors like the contributor’s motivation, level of detail, or time spent on the platform — none of which are captured in the dataset. Because the reason for missingness is tied to unobserved characteristics of the contributor (rather than to other observed columns), this qualifies as NMAR.

To make a stronger case, it would be helpful to have additional data such as the contributor’s activity level or whether they’ve submitted other recipes with or without descriptions. Such data could potentially justify reclassifying the missingness as MAR, but it is not available in this dataset.



### Missingness Dependency Analysis

To analyze missingness patterns, I focused on the `description` column and examined whether its missingness depends on other observed variables.

I performed permutation tests to assess:

1. Whether the missingness of `description` **depends on** the number of ingredients (`n_ingredients`)
2. Whether it **does not depend on** the number of steps (`n_steps`)

<iframe
  src="assets/description_missing_vs_ingredients.html"
  width="700"
  height="500"
  frameborder="0"
></iframe>

**Permutation Test: Description Missingness vs. Number of Ingredients**  
Recipes **with** a description had on average more ingredients than those **without** a description. The resulting p-value was **0.001**, suggesting the missingness of `description` **does** depend on `n_ingredients`.

<iframe
  src="assets/description_missing_vs_steps.html"
  width="700"
  height="500"
  frameborder="0"
></iframe>

**Permutation Test: Description Missingness vs. Number of Steps**  
Here, the p-value was **0.62**, indicating no significant difference in step count between recipes with and without a description. Therefore, we conclude that missingness in `description` **does not** depend on `n_steps`.

These tests provide evidence that some observed features help explain missingness patterns — useful context when deciding how to handle missing values.

## Hypothesis Testing

To investigate whether “healthy”-labeled recipes are genuinely healthier than those without such labels, I conducted a permutation test comparing the average `healthy_index` for the two groups.

### Hypotheses

- **Null Hypothesis (H₀):** The mean `healthy_index` of recipes labeled as "healthy" is equal to that of recipes without the label. Any observed difference is due to random variation.
- **Alternative Hypothesis (H₁):** Recipes labeled as "healthy" have a different mean `healthy_index` than those not labeled "healthy".

This is a two-sided test because I want to detect any difference in either direction — whether the label corresponds to being healthier or unhealthier.

- **Test Statistic:** Absolute difference in group means.
- **Significance Level:** α = 0.05

### Results

- **Mean Healthy Index (Healthy-labeled recipes):** -0.249
- **Mean Healthy Index (Not labeled):** -0.192
- **Observed Difference:** 0.057
- **p-value:** 0.002

Since the p-value is less than 0.05, we reject the null hypothesis. There is statistically significant evidence that the mean `healthy_index` differs between the two groups.

<iframe
  src="assets/healthy_label_vs_index_perm_test.html"
  width="800"
  height="550"
  frameborder="0"
></iframe>

### Interpretation

Although labeled recipes tend to have slightly better (less negative) healthy index values, the overlap in distributions suggests these labels are imperfect proxies for nutritional quality. While the difference is statistically significant, its practical significance is debatable. This reinforces the motivation for creating a more data-driven health indicator.

## Framing a Prediction Problem

The prediction task in this project is to estimate the **average user rating** (`average_rating`) of a recipe based on its nutritional attributes, ingredient complexity, and label metadata.

### Problem Type

This is a **regression** problem, as the response variable (`average_rating`) is continuous and ranges from 0 to 5.

### Response Variable

- **`average_rating`**: The average score assigned to each recipe by users on Food.com.

This variable was selected because it reflects user perception and satisfaction, which are critical to evaluating recipe popularity and success.

### Features Used

All features used in the model are available at the **time of posting** (e.g., nutrition facts, tags, and ingredient count). No user reviews or later-submitted ratings are included as features, to ensure proper data leakage prevention.

### Evaluation Metric

- **Metric Chosen:** RMSE (Root Mean Squared Error)

RMSE is a standard metric for regression tasks. It penalizes large errors more heavily than small ones and gives an interpretable estimate of how far off our model’s predictions are from true values (in rating units). I chose RMSE over alternatives like MAE because it is more sensitive to outliers, which helps flag unusually misrated recipes.

This problem fits within the larger theme of this project: understanding how nutritional content and health labels affect both perception and popularity of recipes.


## Baseline Model

### Prediction Task

My prediction task is to estimate the **average user rating** of a recipe based on its nutritional values and label metadata. This is a **regression** problem, since the response variable `average_rating` is continuous (ranging from 0 to 5).

### Features Used

For the baseline model, I selected a small set of features that were both available at the time a recipe was posted and relevant to perceived healthiness and complexity:

- **Quantitative Features:**
  - `calories`
  - `total_fat`
  - `sugar`
  - `protein`
  - `sodium`
  - `n_ingredients`

- **Nominal Features:**
  - `healthy` (binary flag)
  - `low_in_something` (binary flag)

All numeric columns were used as-is. The binary flags were already 0/1 and did not require additional encoding.

### Model Pipeline

I used a `RandomForestRegressor` as the base model and wrapped all transformations into a single sklearn `Pipeline`. I performed an 80/20 train-test split to evaluate generalization.

### Performance

- **Baseline Model RMSE:** 1.078

This value reflects the model’s average prediction error in terms of rating units. Given that the ratings range from 0 to 5, an RMSE slightly above 1 is reasonable but leaves room for improvement. The model captures some structure in the data (e.g., low-nutrient recipes getting lower ratings), but not enough to make highly accurate predictions yet.

This baseline serves as a foundation for building a more expressive model in Step 7, where I’ll engineer new features and tune hyperparameters.

## Final Model

To improve upon the baseline model, I engineered new features and applied advanced transformations, all within a single `Pipeline`, and optimized hyperparameters using `GridSearchCV`.

### Features and Transformations

The final model uses a rich set of quantitative and binary features:

- **Numeric Features:**
  - Nutritional attributes: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `sat_fat`, `carbs`
  - Recipe complexity: `minutes`, `n_ingredients`
  - Health composite: `healthy_index`

- **Categorical Flags (binary):**
  - `healthy`, `low_in_something`, `free_of_something`

To engineer new features and improve modeling capacity, I applied the following:

- **StandardScaler** to normalize all numeric features.
- **Log transform** on `minutes` to reduce skew from long prep times.
- **PolynomialFeatures** of degree 2 on `healthy_index` to capture potential nonlinear effects between perceived healthiness and ratings.
- Binary columns were passed through without encoding since they were already 0/1.

### Modeling and Optimization

I used a **Random Forest Regressor** and optimized its parameters using `GridSearchCV` with 5-fold cross-validation. Specifically, I tuned:

- `max_depth`: Controls how deep the decision trees can grow (complexity vs. overfitting)
- `min_samples_split`: Controls the minimum number of samples required to split a node (prevents overly specific splits)

### Results

- **Best Parameters:** `{'regressor__max_depth': 5, 'regressor__min_samples_split': 2}`
- **Final Model RMSE:** **1.0765**
- **Baseline RMSE (for comparison):** 1.0783

### Conclusion

Although the RMSE improvement is small, it reflects better handling of nonlinear relationships and data scale, especially through the log and polynomial transformations. This model will serve as the basis for the fairness analysis in the next step.

## Fairness Analysis

To evaluate whether my final model performs fairly across different groups, I conducted a fairness analysis by comparing its RMSE on two recipe groups:

- **Group X (Healthy-labeled)**: Recipes tagged with `"healthy"` in the name or tags
- **Group Y (Non-healthy-labeled)**: Recipes without the `"healthy"` label

This test checks whether my model systematically predicts worse for one group over another.

### Hypotheses

- **Null Hypothesis (H₀):** The model performs equally well for both groups. The difference in RMSE is due to random chance.
- **Alternative Hypothesis (H₁):** The model performs worse (i.e., higher RMSE) for one group compared to the other.

### Evaluation Metric

- **Metric Used:** RMSE (Root Mean Squared Error)
- This is appropriate for my regression task, where we’re predicting a continuous outcome (average rating).

### Observed Results

| Group        | RMSE    |
|--------------|---------|
| Healthy      | 1.0651  |
| Non-Healthy  | 1.0793  |
| **Difference** | **0.0142** |

I performed a permutation test with 1000 resamples to determine if this observed difference is statistically significant.

- **p-value:** 0.6430

### Conclusion

Since the p-value is much greater than 0.05, I fail to reject the null hypothesis. This means that **there is no statistically significant evidence** that the model performs worse for one group compared to the other. Therefore, I conclude that the model is reasonably fair with respect to this health label.

<iframe
  src="assets/fairness_healthy_vs_nonhealthy_rmse_test.html"
  width="800"
  height="550"
  frameborder="0"
></iframe>
