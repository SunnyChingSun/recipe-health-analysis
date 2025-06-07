# "Beyond the Label: Are 'Healthy' Recipes Really Healthy?"
### To what extent do semantic health labels (e.g., “low fat”, “gluten free”, “healthy”) embedded in recipe names and tags correspond to quantified nutritional values, and how do these discrepancies influence user-assigned ratings?

by Sunny Sun (chs019@ucsd.edu)

## Introduction

I like food and I like cooking, and I’m also really into staying on a healthy diet—so I’m pretty familiar with nutrition data. That’s why I’ve always wondered if recipes labeled as “healthy” actually live up to their name. Do these labels reflect real nutritional value, or are they just used to make recipes sound better? In this project, I explore that question using the `Recipes` and `Ratings` dataset from Food.com.

My main research question is: **Are recipes labeled as “healthy” actually healthier based on nutrition facts, and does that label influence user ratings?** 

This question is especially relevant for consumers who rely on health labels to make dietary decisions. If labels like "healthy" or "low fat" are used inconsistently or inaccurately, they may mislead users into making choices that don't align with their health goals. For recipe platform developers and data scientists, understanding this gap could inform improvements to tagging systems and recommender algorithms.

### Dataset
**`Recipes` dataset has 83782 rows where each row indicates a unique recipe and contains the following 12 columns:**

| Column         | Description                                                                                                                                       |
|:---------------|:--------------------------------------------------------------------------------------------------------------------------------------------------|
| `'name'`        | Recipe name                                                                                                                                       |
| `'id'`             | Recipe ID                                                                                                                                         |
| `'minutes'`        | Minutes to prepare recipe                                                                                                                         |
| `'contributor_id'` | User ID who submitted this recipe                                                                                                                 |
| `'submitted'`      | Date recipe was submitted                                                                                                                         |
| `'tags'`          | Food.com tags for recipe                                                                                                                          |
| `'nutrition'`      | Nutrition information in the form [calories (#), total fat (PDV), sugar (PDV), sodium (PDV), protein (PDV), saturated fat (PDV), carbohydrates (PDV)]; PDV stands for "percentage of daily value" |
| `'n_steps'`        | Number of steps in recipe                                                                                                                         |
| `'steps'`          | Text for recipe steps, in order                                                                                                                   |
| `'description'`    | User-provided description                                                                                                                         |

**`Ratings` dataset has 731927 rows where each row indicates a unique review for a recipe and contains the following 5 columns:**

| Column    | Description         |
|:----------|:--------------------|
| `'user_id'`   | User ID             |
| `'recipe_id'` | Recipe ID           |
| `'date'`      | Date of interaction |
| `'rating'`    | Rating given        |
| `'review'`    | Review text         |

To investigate whether the tags truely reflects the healthiness of the recipe and it's relation to ratings, I parsed the nutrition coloum into 7 nutrients: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `sat_fat`, and `carbs`. Then I created a quantitative `Healthy Index` from nutrition data, and compared it to the presence of health-related labels in recipe names and tags. Thus, some key columns from the dataset include:
- `tags`: list of descriptive labels (e.g. “gluten free”, “healthy”)
- `nutrition`: [calories (#), total fat (PDV), sugar (PDV), sodium (PDV), protein (PDV), saturated fat (PDV), carbohydrates (PDV)]
- `name`: recipe title
- `rating`: will turned to `average_rating` (0–5)
- `submitted`: Date recipe was submitted


## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

To begin my analysis, I performed several key data cleaning steps to prepare the dataset for analysis. The original data came in two separate CSV files: `RAW_recipes.csv`, which contained recipe metadata, and `RAW_interactions.csv`, which contained user-submitted ratings. These were merged and cleaned according to the project instructions and additional logic informed by the data structure.

**Cleaning and Processing Steps:**

1. **Merged Datasets:**  
   I performed a left merge of the recipes and interactions datasets on the `id` column to include rating information for each recipe. This ensured that every recipe retained its original metadata, even if it had no ratings.

2. **Handled Zero Ratings:**  
   In the `interactions` data, ratings with a value of `0` are likely placeholders or missing entries. Since 0 is not a valid rating on the Food.com scale (1–5), I replaced them with `np.nan`. This helps avoid skewing averages downward.

3. **Computed Average Ratings:**  
   I grouped ratings by recipe and calculated an `average_rating` per recipe, then merged this Series back into the main `recipes` DataFrame for easier analysis.

4. **Parsed the `nutrition` Column:**  
   The original `nutrition` column was a stringified list containing 7 nutrition facts per recipe. I converted it into individual numeric columns:  
   `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `sat_fat`, and `carbs`.

5. **Normalized Nutrition Values:**  
   Since serving sizes are inconsistent and not explicitly listed, I normalized all nutrient values *per 100 calories*. This helped standardize comparisons between recipes of different portion sizes.

6. **Generated Health Labels:**  
   I created binary indicators for whether a recipe was labeled as:
   - `"healthy"`
   - `"low in something"` (e.g., “low fat”, “low sodium”)
   - `"free of something"` (e.g., “gluten free”, “sugar free”)  
   These were inferred by checking the presence of keywords in both the `tags` and `name` fields.

7. **Created a `healthy_index`:**  
   To quantify the overall nutrition profile of each recipe, I constructed a custom `healthy_index` based on nutritional guidelines:
   ```python
   healthy_index = (
       -0.015 * total_fat - 0.015 * sugar - 0.010 * sat_fat - 0.010 * carbs + 0.035 * protein
   )
   ```
    Sodium was excluded from the index because many recipes only list "salt" as an ingredient, which results in outlier sodium values that are not always meaningful.


The cleanned `recipes` dataframe contains 83756 rows and 27 columns where each rows indicates a unique recipe with average ratings. Since there are too many columns in the dataframe, here only showing the preview of the cleaned dataframe with some key columns:

| name                                 |   healthy_index |   average_rating | healthy   | low_in_something   | free_of_something   |
|:-------------------------------------|----------------:|-----------------:|:----------|:-------------------|:--------------------|
| 1 brownies in the world    best ever |     -0.755058   |                4 | False     | False              | False               |
| 1 in canada chocolate chip cookies   |     -0.700723   |                5 | False     | False              | False               |
| 412 broccoli casserole               |     -0.00513347 |                5 | False     | False              | False               |
| millionaire pound cake               |     -0.769099   |                5 | False     | True               | False               |
| 2000 meatloaf                        |     -0.0430712  |                5 | False     | False              | False               |



### Univariate Analysis

The plot below shows the distribution of `average_rating`. Ratings tend to skew toward the high end, with most falling between 4 and 5. This suggests that users are generous with ratings or that only well-liked recipes accumulate many ratings.

<iframe
  src="assets/univariate_rating_distribution.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>



### Bivariate Analysis
#### Box Plot: Rating by Label Type 
The following boxplot shows how the presence of health-related labels affects user ratings. Recipes labeled as "free_of_something" have slightly **lower median ratings**, but the overall distribution overlaps heavily with unlabeled recipes. This suggests that health labels do **not strongly affect ratings**, although there may be subtle biases in perception or recipe complexity.

<iframe
  src="assets/rating_by_label_boxplot.html"
  width="800"
  height="550"
  frameborder="0"
></iframe>

#### Line Plot: Health Labels Nutritional Trends
Here is a line plot showing the average `healthy_index` over time for different label categories. Most health-labeled groups remained **below the overall average** in healthy index, particularly `gluten_free` and `free_of_something`. Surprisingly, some health-labeled recipes are **not significantly healthier** than unlabeled ones, especially in recent years. This reinforces concerns about the **accuracy of health labels**.

<iframe
  src="assets/healthy_index_trend_by_year.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

#### Scatter Plot: Healthy Index vs Ratings Relationship
This scatter plot explores the relationship between `healthy_index` and `average_rating`. There is **no strong correlation** between nutritional health and user ratings. The trendline is nearly flat, and both healthy-labeled and unlabeled recipes are spread across the entire rating range. This suggests users do not consistently reward healthier recipes with higher ratings.

<iframe
  src="assets/healthy_index_vs_rating_scatter.html"
  width="800"
  height="550"
  frameborder="0"
></iframe>



### Interesting Aggregates

To better understand how health-related labels are used together, I computed the **percentage overlap** between each pair of label categories.

The table below shows, for example, that **45.9%** of recipes labeled `"healthy"` are also labeled `"low_in_something"`, while only **2.5%** of `"gluten_free"` recipes overlap with `"low_in_something"`.

|                   | low_in_something    | low_fat             | low_carb           | healthy           | gluten_free       |
|:------------------|:--------------------|:--------------------|:-------------------|:------------------|:------------------|
| healthy           | 14564/31727 (45.9%) | 10884/17363 (62.7%) | 5069/26136 (19.4%) | nan               | nan               |
| free_of_something | 1687/32837 (5.1%)   | 858/15622 (5.5%)    | 736/18702 (3.9%)   | 1086/19899 (5.5%) | 2301/4677 (49.2%) |
| gluten_free       | 800/31484 (2.5%)    | 359/13881 (2.6%)    | 390/16808 (2.3%)   | 443/18302 (2.4%)  | nan               |
| low_fat           | 11871/29915 (39.7%) | nan                 | nan                | nan               | nan               |
| low_carb          | 14829/29915 (49.6%) | 4751/21949 (21.6%)  | nan                | nan               | nan               |

This shows that:
- Some health tags are often used together (e.g. `"healthy"` + `"low_fat"`),
- Others are more specialized or niche (e.g. `"gluten_free"`),
- And overall, labels are **not applied in a consistent or structured way**, which weakens their reliability.

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
Recipes **with** a description had on average more ingredients than those **without** a description. The resulting p-value was **0.004**, suggesting the missingness of `description` **does** depend on `n_ingredients`.

<iframe
  src="assets/description_missing_vs_steps.html"
  width="700"
  height="500"
  frameborder="0"
></iframe>

**Permutation Test: Description Missingness vs. Number of Steps**  
Here, the p-value was **1.000**, indicating no significant difference in step count between recipes with and without a description. Therefore, we conclude that missingness in `description` **does not** depend on `n_steps`.

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
